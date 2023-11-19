# 3D Tutorial - Step 10 - Enemies
by Monstrous Software


# Step 10 - Enemies

Up to now, the game has been very peaceful.  It is time to introduce some enemy characters: the disgruntled cooks that will throw pans.  The pans will hurt the player if there is a collision.
The player can use the balls to shoot at the cooks.


## Enemy characters


Add an attribute to the game object type class to identify enemy characters: isEnemy. We also add boolean attributes for enemy bullets or friendly bullets.

```java
        public class GameObjectType {
            ...
            public String typeName;
            public boolean isStatic;
            public boolean isPlayer;
            public boolean canPickup;
            public boolean isEnemy;     // <---- new
            public boolean isFriendlyBullet; // <---- new
            public boolean isEnemyBullet; // <---- new
        
            public GameObjectType(String typeName, boolean isStatic, boolean isPlayer, boolean canPickup, boolean isEnemy, boolean isFriendlyBullet, boolean isEnemyBullet) {
                this.typeName = typeName;
                this.isStatic = isStatic;
                this.isPlayer = isPlayer;
                this.canPickup = canPickup;
                this.isEnemy = isEnemy;
                this.isFriendlyBullet = isFriendlyBullet;
                this.isEnemyBullet = isEnemyBullet;
            }
        }
```

Add a new type 'typeEnemy' to list of type constants.  Note how all the constructor calls need three extra boolean parameters compared to the previous step.

```java
       public class GameObjectType {

            public final static GameObjectType TYPE_STATIC = new GameObjectType("static", true, false, false, false, false, false);
            public final static GameObjectType TYPE_PLAYER = new GameObjectType("player", false, true, false, false, false, false);
            public final static GameObjectType TYPE_PICKUP_COIN = new GameObjectType("coin", false, false, true, false , false, false);
            public final static GameObjectType TYPE_PICKUP_HEALTH = new GameObjectType("healthpack", false, false, true, false , false, false);
            public final static GameObjectType TYPE_DYNAMIC = new GameObjectType("dynamic", false, false, false, false, false, false);
            public final static GameObjectType TYPE_ENEMY = new GameObjectType("enemy", false, false, false, true, false, false);
            public final static GameObjectType TYPE_FRIENDLY_BULLET = new GameObjectType("bullet", false, false, false, false, true,false);
            public final static GameObjectType TYPE_ENEMY_BULLET = new GameObjectType("bullet", false, false, false, false,false, true);

            ...
        }
```

Now we can add some enemies in World.populate():

```java
        private void populate() {
            ...
            /// add enemies
            world.spawnObject(GameObjectType.TYPE_ENEMY, "cook",  "cookProxy", CollisionShapeType.CAPSULE, true, new Vector3(-15, 1f, -18), Settings.playerMass  );  // bad guy
            world.spawnObject(GameObjectType.TYPE_ENEMY, "cook",  "cookProxy", CollisionShapeType.CAPSULE, true, new Vector3(15, 1f, 18), Settings.playerMass  );  // bad guy
            world.spawnObject(GameObjectType.TYPE_ENEMY, "cook",  "cookProxy", CollisionShapeType.CAPSULE, true, new Vector3(-25, 1f, 25), Settings.playerMass  );  // bad guy
            world.spawnObject(GameObjectType.TYPE_ENEMY, "cook",  "cookProxy", CollisionShapeType.CAPSULE, true, new Vector3(25, 1f, 25), Settings.playerMass  );  // bad guy
            //...
        }
```

If we run the program now we can see four ominous characters have been added. But they are still completely static. Some enemy behaviour is needed.


## Behaviour

We'll use a Behaviour class to define the behaviour of the different object types in the world.  For example, how enemy characters behave.
The Behaviour class sets a general template, we will subclass this for our enemy character.  The Behaviour class also defines a factory method to create
a Behaviour instance of the correct subclass for the given game object.   For now, we only have a subclass CookBehaviour for the one type of enemy we have.
All other game objects get a 'null' Behaviour instance. 

```java
        public class Behaviour {
            protected GameObject go;
        
            protected Behaviour(GameObject go) {
                this.go = go;
            }
        
            public void update(World world, float deltaTime ) { }
        
            // factory for Behaviour instance depending on object type
            public static Behaviour createBehaviour(GameObject go){
                if(go.type.isEnemy)
                    return new CookBehaviour(go);
                return null;
            }
        }
```

Now to subclass Behaviour for the enemy cook character.  The cook will also use a capsule collision shape just like the player and we will also disable any rotation of the capsule.  For this, we call
the method `setCapsuleCharacteristics()` which was formerly known as `setPlayerCharacteristics()`. The update() method will be called each frame and allows the enemy character to 
act out some behaviour pattern.  In this case, it will be very basic behaviour.  It stops doing anything, when the character is dead (health <= 0). (We will add this field shortly).
Otherwise, it will always turn to face the player character and move towards the player up to some distance.  Then every so often it will spawn an enemy bullet (a pan) which is thrown in the direction of 
the player.  The shootPan() method is very similar to the shootBall() method we saw earlier.

```java
        public class CookBehaviour extends Behaviour {
        
            private static final float SHOOT_INTERVAL = 2f;     // seconds between shots
        
            private float shootTimer;
            private Vector3 spawnPos = new Vector3();
            private Vector3 shootDirection = new Vector3();
            private Vector3 direction = new Vector3();
            private Vector3 targetDirection = new Vector3();
            private Vector3 angularVelocity = new Vector3();
        
            public CookBehaviour(GameObject go) {
                super(go);
                shootTimer = SHOOT_INTERVAL;
                go.body.setCapsuleCharacteristics();
            }
        
            public Vector3 getDirection() {
                return direction;
            }
        
            @Override
            public void update(World world, float deltaTime ) {
                if(go.health <= 0)   // don't do anything when dead
                    return;
        
                // move towards player
                targetDirection.set(world.getPlayer().getPosition()).sub(go.getPosition());  // vector towards player
                targetDirection.y = 0;  // consider only vector in horizontal plane
                float distance = targetDirection.len();
                targetDirection.nor();      // make unit vector
                direction.set(targetDirection);
                if(distance > 5f)   // move unless quite close
                    go.body.applyForce(targetDirection.scl(3f));
        
        
                // rotate to follow player
                angularVelocity.set(0,0,0);
                targetDirection.nor();      // make unit vector
                Vector3 facing = go.getDirection();                                             // vector we're facing now
                float dot = targetDirection.dot(facing);                                        // dot product = cos of angle between the vectors
                float cross = Math.signum(targetDirection.crs(facing).y);                       // cross product to give direction to turn
                if(dot < 0.99f)                         // if not facing player
                    angularVelocity.y = -cross;         // turn towards player
                go.body.applyTorque(angularVelocity);
        
                // every so often shoot a pan
                shootTimer -= deltaTime;
                if(shootTimer <= 0 && distance < 20f && world.getPlayer().health > 0) {
                    shootTimer = SHOOT_INTERVAL;
                    shootPan(world);
                }
            }
        
            private void shootPan(World world) {
                spawnPos.set(direction);
                spawnPos.nor().scl(1f);
                spawnPos.add(go.getPosition()); // spawn from 1 unit in front of the character
                spawnPos.y += 1f;
                GameObject pan = world.spawnObject(GameObjectType.TYPE_ENEMY_BULLET, "pan", "panProxy", CollisionShapeType.MESH, true, spawnPos, Settings.panMass );
                shootDirection.set(direction);        // shoot forward
                shootDirection.y += 0.5f;       // and slightly up
                shootDirection.scl(Settings.panForce);   // scale for speed
                pan.body.geom.getBody().setDamping(0.0f, 0.0f);
                pan.body.applyForce(shootDirection);
                pan.body.applyTorque(Vector3.Y);    // add some spin
            }
        }
```



Let use extend the GameObject class to give each object a health value, which will range from 0 to 1, and a Behaviour instance. We also add an update() method to GameObject
which will in turn calls Behaviour.update() unless the behaviour is null. 

```java        
        public class GameObject {
        
            public final GameObjectType type;
            public final Scene scene;
            public final PhysicsBody body;
            public final Vector3 direction;
            public boolean visible;
            public float health;
            public Behaviour behaviour;
        
            public GameObject(GameObjectType type, Scene scene, PhysicsBody body) {
                this.type = type;
                this.scene = scene;
                this.body = body;
                body.geom.setData(this);            // the geom has user data to link back to GameObject for collision handling
                visible = true;
                direction = new Vector3();
                health = 1f;
                behaviour = Behaviour.createBehaviour(this);
            }
                              
            public void update(World world, float deltaTime ){
                if(behaviour != null)
                    behaviour.update(world, deltaTime);
            }
            
            public boolean isDead() {
                return health <= 0;
            }
            //...
        }
```

For the cook character we will also use a capsule for collision geometry, and we will use the same trick as for the player to keep the capsule upright, namely to disable angular rotation 
on the physics body.  To rotate the modelInstance in the game view, we will use the direction vector from the CookBehaviour class to find the forward facing direction.
We can also now use the game object type to determine how we should update
the modelInstance transform.  For the player we get the orientation from the player controller, for the enemy
we get it from the CookBehaviour object and for other game objects we get it from the rigid body orientation.


```java  
        private void syncToPhysics() {
            for(GameObject go : gameObjects){
                if( go.body.geom.getBody() != null) {
                    if(go.type == GameObjectType.TYPE_PLAYER){
                        // use information from the player controller, since the rigid body is not rotated.
                        player.scene.modelInstance.transform.setToRotation(Vector3.Z, playerController.getForwardDirection());
                        player.scene.modelInstance.transform.setTranslation(go.body.getPosition());
                    }
                    else if(go.type == GameObjectType.TYPE_ENEMY){
                        CookBehaviour cb = (CookBehaviour) go.behaviour;
                        go.scene.modelInstance.transform.setToRotation(Vector3.Z, cb.getDirection());
                        go.scene.modelInstance.transform.setTranslation(go.body.getPosition());
                    }
                    else
                        go.scene.modelInstance.transform.set(go.body.getPosition(), go.body.getOrientation());
                }
            }
        }
```

In the update() method of the World class, call the new `update()` method for each game object:

```java 
        public void update( float deltaTime ) {
            playerController.update(player, deltaTime);
            physicsWorld.update();
            syncToPhysics();
            for(GameObject go : gameObjects)            
                go.update(this, deltaTime);
        }
```


To keep score during the game, let's add a class with game statistics, e.g. the number of coins collected and the number of enemies remaining.
The World class with have a GameStats field called `stats`.

```java
        public class GameStats {
            public float gameTime;
            public int numCoins;
            public int coinsCollected;
            public int numEnemies;
            public boolean levelComplete;
        
            public GameStats() {
                reset();
            }
        
            public void reset() {
                gameTime = 0;
                numCoins = 0;
                coinsCollected = 0;
                numEnemies = 0;
                levelComplete = false;
            }
        }
```

Now that we have bullets flying around we also need to handle when those bullets hit something relevant. Let us modify handleCollision()
which we introduced in the previous step:

```java
        private void handleCollision(GameObject go1, GameObject go2){
            if(go1.type.isPlayer && go2.type.canPickup)
                pickup(go1, go2);

            if(go1.type.isPlayer && go2.type.isEnemyBullet)     //<-- new
                bulletHit(go1, go2);                            //<-- new

            if(go1.type.isEnemy && go2.type.isFriendlyBullet)   //<-- new
                bulletHit(go1, go2);                            //<-- new
       }
```

The pickup() method is adapted to count the number of coins picked up or to add some health depending on the type of object picked up:

```java
        private void pickup(GameObject character, GameObject pickup){
            removeObject(pickup);
            if(pickup.type == GameObjectType.TYPE_PICKUP_COIN)
                stats.coinsCollected++;
            if(pickup.type == GameObjectType.TYPE_PICKUP_HEALTH)
                character.health = Math.min(character.health + 0.5f, 1f);   // +50% health
        }
```

The new method bulletHit() is called when a character (the player or an enemy) is hit by a bullet from the opposing side.
It takes 25% health and removes the character when dead.

```java
        private void bulletHit(GameObject character, GameObject bullet) {
            removeObject(bullet);
            character.health -= 0.25f;      // - 25% health
            if(character.isDead()) 
                removeObject(character);
        }
```

The shootBall() method needs a bit of updating, because the spawnObject() method has more parameters now. We also add a check to stop the 
player from shooting once they're dead:

```java
        public void shootBall() {
            if(player.isDead())
                return;
            dir.set( playerController.getViewingDirection() );
            spawnPos.set(dir);
            spawnPos.add(player.getPosition()); // spawn from 1 unit in front of the player
            GameObject ball = spawnObject(GameObjectType.TYPE_FRIENDLY_BULLET, "ball", null, CollisionShapeType.SPHERE, true, spawnPos, Settings.ballMass );
            shootDirection.set(dir);        // shoot forward
            shootDirection.y += 0.5f;       // and slightly up
            shootDirection.scl(Settings.ballForce);   // scale for speed
            ball.body.geom.getBody().setDamping(0.0f, 0.0f);
            ball.body.applyForce(shootDirection);
        }
```

Since we're now also removing game objects when they have been shot, we need to update the `remove()` method. We update the statistics for number of enemies remaining.
We also call a new `GameObject.dispose()` method which will take care that the ODE rigid body and collision geometry are properly destroyed.

```java
        public void removeObject(GameObject gameObject){
            gameObject.health = 0;
            if(gameObject.type == GameObjectType.TYPE_ENEMY)
                stats.numEnemies--;
            gameObjects.removeValue(gameObject, true);
            gameObject.dispose();
            isDirty = true;     // list of game objects has changed
        }
```

Where `GameObject.destroy()` is defined as follows:

```java
        @Override
        public void dispose() {
            body.destroy();
        }
```

And `PhysicsBody.destroy()` as follows:

```java
        public void destroy() {
            if(geom.getBody() != null)
                geom.getBody().destroy();
            geom.destroy();
        }
```

This concludes step 10.  We've introduced an enemy character, the concept of health points and taking damage from each other's projectiles and using health packs to restore health. 

![battle.png](images%2Fbattle.png)

-- action shot
