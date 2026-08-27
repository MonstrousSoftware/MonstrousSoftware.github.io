# 3D Tutorial - Step 5 - Physics and collisions
by Monstrous Software


# Step 5 - Physics and collisions

As we saw in the previous step we don't have collision detection yet, nor do we have any physics (rigid body simulation).  
We will add both by making use of a physics library.  

There are multiple physics libraries available, such as Bullet, ODE, Jolt, PhysX and since recently Box3d.
One issue with these libraries that for speed they are usually coded in C or C++ and provided as a native library for each platform.
This means we need a Java wrapper to call the native library via JNI which makes the app less portable, particularly to a web version.

ODE on the other hand has been fully translated to Java in the ODE4j project, and we will use this for our tutorial.  Despite being pure Java, it still won't run on the web (TeaVM) version,
for example because it makes use of multi-threading.  There was a derivation called gdx-ode4j by AntzGames which worked on the web version
but this is no longer maintained.

This means we will continue this tutorial focusing on the desktop version only. If you try to build the web version, you will get many errors.

The manual for ODE can be found at [ode.org](https://ode.org/wiki/index.php/Manual).

Open the file `build.gradle` in the `core` module of our project. This file defines what dependencies the core module of our project has on external libraries. 
Add a line to the `dependencies` block to include the latest version of ODE4j:

```java
        dependencies {
            api "com.badlogicgames.gdx-controllers:gdx-controllers-core:$gdxControllersVersion"
            api "com.badlogicgames.gdx:gdx:$gdxVersion"
            implementation "org.ode4j:core:$ode4jVersion"                   // <------ add this line
            api "com.github.mgsx-dev.gdx-gltf:gltf:$gdxGltfVersion"                 
            api "de.golfgl.gdxcontrollerutils:gdx-controllerutils-mapping:$controllerMappingVersion"
            api "de.golfgl.gdxcontrollerutils:gdx-controllerutils-scene2d:$controllerScene2DVersion"

            if(enableGraalNative == 'true') {
                implementation "io.github.berstanio:gdx-svmhelper-annotations:$graalHelperVersion"
            }
        }
```
And add the following line to `gradle.properties` in the project's root directory. This file is typically where
we keep track of dependency versions:

```
    ode4jVersion=0.5.4
```

As we have updated a gradle file, we now need to refresh the Gradle project. Click on the Gradle refresh icon in the IDE:

![gradle refresh button](/assets/images/gradle-refresh.png)




We will wrap most of the ODE4j specifics inside a new class called PhysicsWorld:
```java
   public class PhysicsWorld implements Disposable {
        static final float TIME_STEP = 0.025f;  // fixed physics time step
    
        DWorld world;
        public DSpace space;
        private final DJointGroup contactGroup;
        private float timeElapsed;
    
        public PhysicsWorld() {
            OdeHelper.initODE2(0);
            Gdx.app.log("ODE version", OdeHelper.getVersion());
            Gdx.app.log("ODE config", OdeHelper.getConfiguration());
            contactGroup = OdeHelper.createJointGroup();
            reset();
        }
    
        // reset world, note this invalidates (orphans) all rigid bodies and geoms so should be used in combination with deleting all game objects
        public void reset() {
            if(world != null)
                world.destroy();
            if(space != null)
                space.destroy();
    
            world = OdeHelper.createWorld();
            space = OdeHelper.createSapSpace( null, DSapSpace.AXES.XZY  );
    
            world.setGravity (0, Settings.gravity, 0);
            world.setCFM (1e-5);
            world.setERP (0.4);
            world.setQuickStepNumIterations (40);
            world.setAngularDamping(0.5f);
    
            // set auto disable parameters to make inactive objects go to sleep
            world.setAutoDisableFlag(true);
            world.setAutoDisableLinearThreshold(0.1);
            world.setAutoDisableAngularThreshold(0.1);
            world.setAutoDisableTime(2);
            timeElapsed = 0;
        }
    
        // update the physics with fixed time steps
        public void update(float deltaTime) {
            timeElapsed += deltaTime;
            while(timeElapsed > TIME_STEP) {
                space.collide(null, nearCallback);
                world.quickStep(TIME_STEP);
                contactGroup.empty();
                timeElapsed -= TIME_STEP;
            }
        }
    
        // called for potential collisions
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
    
                    for (int i = 0; i < n; i++) {
                        DContact contact = contacts.get(i);
                        contact.surface.mode = dContactBounce | dContactSoftCFM;
                        contact.surface.mu = 0.1;   // friction coefficient
                        contact.surface.bounce = 0.9;       // Set restitution (0.0 to 1.0)
                        contact.surface.bounce_vel = 0.01;   // Minimum velocity to trigger bounce
                        contact.surface.soft_cfm = 0.001;
    
                        DJoint c = OdeHelper.createContactJoint(world, contactGroup, contact);
                        c.attach(o1.getBody(), o2.getBody());
                    }
                }
            }
        };
    
        @Override
        public void dispose() {
            contactGroup.destroy();
            space.destroy();
            world.destroy();
            OdeHelper.closeODE();
        }
    }   
```
## ODE Physics intro

ODE supports rigid bodies (bodies) and geometry nodes (geoms). A rigid body is a point mass that has mass, has a position and may have a linear velocity and an angular (rotational) velocity. 
Bodies can be accelerated by applying a force on them and can be rotated by applying torque on them.  By default, all bodies have gravity applied to them. Geoms are shapes that are used by ODE for collision detection.   
There are some basic shapes available: boxes, spheres, capsules, cylinders, etcetera. 
Or you can define a geom shape via a mesh to make any shape you want.  For efficiency, it is best to use the simplest shapes you can.
Rigid bodies live in a container of type DWorld amd geoms live in a container called DSpace.
Often we will link a rigid body and a geom.  Static game objects, like the floor for example, may have only a geom for collisions, but no rigid body as it will never move.  
Rigid bodies can be connected by joints.  There are different types of joints available. For example, if you want to model a car you may want to connect the wheels with joints to the chassis. You may also want to use
joints to simulate the car suspension.

ODE also uses temporary joints to simulate collision response. Such joints will only exist for one step of the simulation.  They are created for every collision that is detected to join the two colliding bodies with a contact joint. 
Then in the physics simulation step these joints will push the bodies apart depending on the bounciness, the friction parameters, etcetera.

The main simulation loop is the following:
```java
        space.collide(null, nearCallback);
        world.quickStep(TIME_STEP);
        contactgroup.empty();
```


The first line will test for collisions of all the geoms in the space. For any pair of potential colliding geoms, the callback function is called. The callback function will check if the shapes are indeed colliding and if so create a contact joint for each contact between the two geoms.
The second line will advance all the rigid bodies for one time step, applying all the velocities, forces and torques to calculate their new position, orientation and velocities.  
ODE works best with a fixed time step, so we don't use deltaTime here, but we iterate to do as many fixed time steps as needed since the previous frame. 
The third line deletes all the contact joints that we created in the collision step, ready for the next loop.

We will need to copy the new positions and orientations to the game objects before rendering. This will happen in `World#update`.

For the gravity value we refer to a new variable in the Settings class:
```java
        static public float gravity = -9.8f;   // meters/s^2
```

## Update World class

The PhysicsWorld class will be instantiated in the World class. Add the following variable to the World class:
```java
        public final PhysicsWorld physicsWorld;
```
Initialize it in the constructor before the call of populate():
```java   
        physicsWorld = new PhysicsWorld();
```

Perform a reset of the physics world in the `clear` method.
```java   
    public void clear() {}
        physicsWorld.reset();
        //...
    }
```

Create an update method in World that calls the update method of the physics world and then copies
the positions and orientations of the rigid bodies to the model instances:
```java
        public void update( float deltaTime ) {
            physicsWorld.update();
            for(GameObject go : gameObjects){
                if( go.body.geom.getBody() != null) {
                    go.scene.modelInstance.transform.set(go.body.getPosition(), go.body.getBodyOrientation());
                }
            }
        }
```
And dispose the physics world instance in World.dispose():
```java
        physicsWorld.dispose();
```

## PhysicsBody

Let us create a class that represents a physics body. Its main purpose is to encapsulate a DGeom object (i.e. a collision shape).  Note that a geom can in turn be linked to a rigid body (a point mass) using `geom.setBody()`.

Then for convenience we add methods to get the position and orientation of the geom, converting from ODE's internal format to the LibGDX standard format for Vector3 and Quaternion.

Lastly, we add a ModelInstance object that we will use in debug mode to visualize the collision shape.  It can be very hard to find errors in the physics handling if we can't see the collision shapes.
So we introduce this debug option early.
```java
        public class PhysicsBody {
        
            public DGeom geom;
            private final Vector3 position;               // for convenience, matches geom.getPosition() but converted to Vector3
            private final Quaternion quaternion;          // for convenience, matches geom.getQuaternion() but converted to LibGDX Quaternion
            public final ModelInstance debugInstance;    // visualisation of collision shape for debug view
        
            public PhysicsBody(DGeom geom, ModelInstance debugInstance) {
                this.geom = geom;
                this.debugInstance = debugInstance;
                position = new Vector3();
                quaternion = new Quaternion();
            }
        
            public Vector3 getPosition() {
                DVector3C pos = geom.getPosition();
                position.x = (float) pos.get0();
                position.y = (float) pos.get1();
                position.z = (float) pos.get2();
                return position;
            }
        
            public void setPosition( Vector3 pos ) {
                geom.setPosition(pos.x, pos.y, pos.z);
                DBody rigidBody = geom.getBody();
                if(rigidBody != null)
                    rigidBody.setPosition(pos.x, pos.y, pos.z);
            }
        
            public Quaternion getOrientation() {
                DQuaternionC odeQ = geom.getQuaternion();
                float ow = (float) odeQ.get0();
                float ox = (float) odeQ.get1();
                float oy = (float) odeQ.get2();
                float oz = (float) odeQ.get3();
                quaternion.set(ox, oy, oz, ow);
                return quaternion;
            }
        
            // get orientation of rigid body, i.e. without any geom offset rotation
            public Quaternion getBodyOrientation() {
                DQuaternionC odeQ;
                if(geom.getBody() == null)      // if geom does not have a body attached, fall back to geom orientation
                    odeQ = geom.getQuaternion();
                else
                    odeQ = geom.getBody().getQuaternion();
                float ow = (float) odeQ.get0();
                float ox = (float) odeQ.get1();
                float oy = (float) odeQ.get2();
                float oz = (float) odeQ.get3();
                quaternion.set(ox, oy, oz, ow);
                return quaternion;
            }
        
            public void setOrientation( Quaternion q ){
                DQuaternion odeQ = new DQuaternion(q.w, q.x, q.y, q.z);       // convert to ODE quaternion
                geom.setQuaternion(odeQ);
                DBody rigidBody = geom.getBody();
                if(rigidBody != null)
                    rigidBody.setQuaternion(odeQ);
            }
        }   
```

To present the debug view of all collision shapes corresponding to World game objects, we add a simple class with a ModelBatch that shows all debug instances.  It will use a color code to show which shapes are static, which are active and which are sleeping. :
Active shapes will fall asleep if they have not moved for a while, which means fewer calculations need to be done, allowing better performance.

```java
    public class PhysicsView implements Disposable {
        // colours to use for active vs. sleeping geoms
        static private final Color COLOR_ACTIVE = Color.GREEN;
        static private final Color COLOR_SLEEPING = Color.TEAL;
        static private final Color COLOR_STATIC = Color.GRAY;
    
        private final ModelBatch modelBatch;
        private final World world;      // reference
    
        public PhysicsView(World world) {
            this.world = world;
            modelBatch = new ModelBatch();
        }
    
        public void render( Camera cam ) {
            modelBatch.begin(cam);
            int num = world.getNumGameObjects();
            for(int i = 0; i < num; i++)
                renderCollisionShape(world.getGameObject(i).body);
            modelBatch.end();
        }
    
    
        private void renderCollisionShape(PhysicsBody body) {
            // move & orient debug modelInstance in line with geom
            body.debugInstance.transform.set(body.getPosition(), body.getBodyOrientation());
    
            // use different colour for static/sleeping/active objects and for active ones
            Color color = COLOR_STATIC;
            if (body.geom.getBody() != null) {
                if (body.geom.getBody().isEnabled())
                    color = COLOR_ACTIVE;
                else
                    color = COLOR_SLEEPING;
            }
            body.debugInstance.materials.first().set(ColorAttribute.createDiffuse(color));   // set material colour
    
            modelBatch.render(body.debugInstance);
        }
    
        @Override
        public void dispose() {
            modelBatch.dispose();
        }
    }
```

## Extension of GameObject class

Now the GameObject class needs to be extended so that each game object not only has a Scene field for the rendering but also a PhysicsBody field for the dynamic movement.
```java
        public class GameObject {
        
            public final Scene scene;
            public final PhysicsBody body;
        
            public GameObject(Scene scene, PhysicsBody body) {
                this.scene = scene;
                this.body = body;
                body.geom.setData(this);            // the geom has user data to link back to GameObject for collision handling
            }
        }
```
Whenever we construct a GameObject, we set the data field of the related geom to the game object itself. This will be handy when we detect collisions between geoms to find the corresponding game objects.
We can use this to pick up a weapon for example.

We'll also need to define a mass for the rigid body and a collision shape for the geom.
The mass determines how much a body accelerates when a force is applied.

The shape of the graphics object, the mesh, is typically approximated with a simplified shape for the sake of efficient collision detection.
This we call the collision proxy or the collision shape. For example, the player may have a capsule as a collision shape.  A wall may have a box shape to approximate it, a skull is perhaps approximated by a sphere.
We can also use the mesh itself as collision shape. This is the most accurate, but is inefficient if the mesh is complex.

For the shape type, we'll introduce an enumeration:
```java
        public enum CollisionShapeType {
            BOX, SPHERE, CAPSULE, CYLINDER, MESH
        }
```

![](/assets/images/collision-proxy.png)

-- collisions shapes visualized

The following class is a factory class for PhysicsBody objects.
We use the bounding box of the node to obtain the object's dimensions. Depending on the desired shape type, it will create a geom of the right type and dimensions (we will add support for the MESH type later).
To visualize the collision shapes it will also create a ModelInstance of the using the ModelBuilder.  If the game object is a dynamic object, it will create a rigid body and attach it to the geom.

```java
        public class PhysicsBodyFactory implements Disposable {

            public static final long CATEGORY_STATIC  = 1;      // collision flags
            public static final long CATEGORY_DYNAMIC  = 2;     // collision flags
        
            private final PhysicsWorld physicsWorld;
            private final DMass massInfo;
            private final Vector3 position;
            private final Quaternion q;
            private final ModelBuilder modelBuilder;
            private final Material material;
            private final Array<Disposable> disposables;
        
            public PhysicsBodyFactory(PhysicsWorld physicsWorld) {
                this.physicsWorld = physicsWorld;
                massInfo = OdeHelper.createMass();
                position = new Vector3();
                q = new Quaternion();
                modelBuilder = new ModelBuilder();
                material = new Material(ColorAttribute.createDiffuse(Color.WHITE));
                disposables = new Array<>();
            }
        
            public PhysicsBody createBody( ModelInstance collisionInstance, CollisionShapeType shapeType, float mass, boolean isStatic) {
                BoundingBox bbox = new BoundingBox();
                DQuaternion rotX90 = DQuaternion.fromEulerDegrees(90, 0, 0);     // rotate 90 degrees around X, i,e, Q = (a, a, 0, 0), a = sqrt(2)/2
                Node node = collisionInstance.nodes.first();
                node.calculateBoundingBox(bbox, false); // bounding box without the transform
                float w = bbox.getWidth();
                float h = bbox.getHeight();
                float d = bbox.getDepth();
        
                DGeom geom;
                ModelInstance instance;
                float diameter = 0;
                float radius = 0;
                float len;
        
                switch(shapeType) {
                    case BOX:
                        geom = OdeHelper.createBox(physicsWorld.space, w, h, d);
                        break;
                    case SPHERE:
                        diameter = Math.max(Math.max(w, h), d);
                        radius = diameter/2f;
                        geom = OdeHelper.createSphere(physicsWorld.space, radius);
                        break;
                    case CAPSULE:
                        diameter = Math.max(w, d);
                        radius = diameter/2f; // radius of the cap
                        len = h - 2*radius;     // height of the cylinder between the two end caps
                        geom = OdeHelper.createCapsule(physicsWorld.space, radius, len);
                        break;
                    case CYLINDER:
                        diameter = Math.max(w, d);
                        radius = diameter/2f; // radius of the cap
                        len = h;     // height of the cylinder between the two end caps
                        geom = OdeHelper.createCylinder(physicsWorld.space, radius, len);
                        break;
                    default:
                        throw new RuntimeException("Unknown shape type");
                }
        
                if(isStatic) {
                    geom.setCategoryBits(CATEGORY_STATIC);   // which category is this object?
                    geom.setCollideBits(0);                  // which categories will it collide with?
                    // note: geom for static object has no rigid body attached
                }
                else {
                    DBody rigidBody = OdeHelper.createBody(physicsWorld.world);
                    massInfo.setBox(1, w, h, d);
                    massInfo.setMass(mass);
                    rigidBody.setMass(massInfo);
                    rigidBody.enable();
                    rigidBody.setAutoDisableDefaults();
                    rigidBody.setGravityMode(true);
                    rigidBody.setDamping(0.01, 0.1);
        
                    geom.setBody(rigidBody);
                    geom.setCategoryBits(CATEGORY_DYNAMIC);
                    geom.setCollideBits(CATEGORY_DYNAMIC|CATEGORY_STATIC);
                    if(shapeType == CollisionShapeType.CYLINDER || shapeType == CollisionShapeType.CAPSULE) {
                        // rotate geom 90 degrees around X because ODE geom cylinders and capsules shapes are created using Z as long axis
                        // and we want the shape to be oriented along the Y axis which is up.
                        geom.setOffsetQuaternion(rotX90);    // set standard rotation from rigid body to geom
                    }
                }
        
        
                // create a debug model matching the collision geom shape
                modelBuilder.begin();
                MeshPartBuilder meshBuilder;
                meshBuilder = modelBuilder.part("part", GL20.GL_LINES, VertexAttributes.Usage.Position , material);
                switch(shapeType) {
                    case BOX:
                        BoxShapeBuilder.build(meshBuilder, w, h, d);
                        break;
                    case SPHERE:
                        SphereShapeBuilder.build(meshBuilder, diameter, diameter, diameter , 8, 8);
                        break;
                    case CAPSULE:
                        CapsuleShapeBuilder.build(meshBuilder, radius, h, 12);
                        break;
                    case CYLINDER:
                        CylinderShapeBuilder.build(meshBuilder, diameter, h, diameter, 12);
                        break;
                }
                Model modelShape = modelBuilder.end();
                disposables.add(modelShape);
                instance = new ModelInstance(modelShape, Vector3.Zero);
        
                PhysicsBody body = new PhysicsBody(geom, instance);
        
                // copy position and orientation from modelInstance to body
                collisionInstance.transform.getTranslation(position);
                collisionInstance.transform.getRotation(q);
                body.setPosition(position);
                body.setOrientation(q);
                return body;
            }
        
            @Override
            public void dispose() {
                for(Disposable d : disposables)
                    d.dispose();
            }
        }
```

There is a little trick with regard to some of the geom shapes. ODE creates cylinder and capsule shapes with the main axis along Z. In LibGDX the shape builders will use Y as main axis.
Therefore, we rotate the geom by 90 degrees around the X axis when we attach it to the rigid body.


## Modeling caveat

Note that there is an important constraint on how the objects are modeled in Blender in order for the collision shapes to be in the correct position:
Objects need to be modeled with their centre on the origin and be axis-aligned.  The point at the origin will be used as centre of mass for the rigid body physics.
However, the objects can then still be moved around in Blender as long as the transforms are not applied.
So in Blender you need to:
1. position the object centred on the origin. 
2. Then use 'apply all transforms' (Control-A).
3. Now you can move the object to where you want it to appear. For example, on top of the ground instead of halfway in the ground. You can also rotate it or scale it.
4. But do NOT apply transforms of this new position, orientation or scale.

This way the collision shape will be calculated relative to the original position which was centred on the origin.
The final transform is taken from node.globalTransform and is used to set the modelInstance transform.

![](/assets/images/origin-centred.png)
<figcaption>step 1 - centre the object on the origin and apply transform.</figcaption>

![](/assets/images/origin-adjusted.png)
<figcaption>step 3 - position the object to the desired location, e.g. with the feet at ground level, but do not apply transform.</figcaption>

![](/assets/images/floating-duck.png)
<figcaption>if the applied transform is not centred on the origin, the collision shape and the graphical mesh will be misaligned. Here the feet are placed at the centre of mass. </figcaption>


## SpawnObject

It is time to adapt the method World.spawnObject.  It now has extra parameters for mass and shape type.  
```java
    public GameObject spawnObject(boolean isStatic, String name, CollisionShapeType shape, Vector3 position, float mass){
        Scene scene = new Scene(sceneAsset.scene, name);
        if(scene.modelInstance.nodes.size == 0)
            throw new RuntimeException("Cannot find node in GLTF file: " + name);
    
        applyNodeTransform(scene.modelInstance, scene.modelInstance.nodes.first());         // incorporate nodes' transform into model instance transform
        scene.modelInstance.transform.translate(position);
    
        PhysicsBody body = factory.createBody(scene.modelInstance, shape, mass, isStatic);
        GameObject go = new GameObject(scene, body);
        gameObjects.add(go);
        isDirty = true;         // list of game objects has changed
        return go;
    }
    
    private void applyNodeTransform(ModelInstance modelInstance, Node node ){
        modelInstance.transform.mul(node.globalTransform);
        node.translation.set(0,0,0);
        node.scale.set(1,1,1);
        node.rotation.idt();
        modelInstance.calculateTransforms();
    }
```

The Populator class needs to be adapted to support the extra parameters required per object, such as isStatic, shape type and mass.
```java
        public class Populator {
        
            public static void populate(World world) {
                world.clear();
        
                world.spawnObject(true, "brickcube", CollisionShapeType.BOX, Vector3.Zero, 1);
                world.spawnObject(true, "groundbox", CollisionShapeType.BOX, Vector3.Zero, 1f);
                world.spawnObject(true, "brickcube.001", CollisionShapeType.BOX,Vector3.Zero, 1f);
                world.spawnObject(true, "brickcube.002", CollisionShapeType.BOX,Vector3.Zero, 1f);
                world.spawnObject(true, "brickcube.003", CollisionShapeType.BOX,Vector3.Zero, 1f);
                world.spawnObject(true, "wall", CollisionShapeType.BOX,Vector3.Zero, 1f);
                world.spawnObject(false, "ball", CollisionShapeType.SPHERE, new Vector3(0,4,0), 1f);
                world.spawnObject(false, "ball", CollisionShapeType.SPHERE,new Vector3(-1,5,0), 1f);
                world.spawnObject(false, "ball", CollisionShapeType.SPHERE, new Vector3(-2,6,0), 1f);
                world.player = world.spawnObject(false, "ducky",CollisionShapeType.CAPSULE, new Vector3(0,1,0), 1f);
            }
        }
```
For testing, it is useful to restart the game from the initial state.  To allow this by pressing 'R', add the following line to `GameScreen.render()`:

```java
        if (Gdx.input.isKeyJustPressed(Input.Keys.R))
            Populator.populate(world);
```
This concludes step 5 of this tutorial. We've now added collision handling and physics to our template.  
You should be able to see the three balls in the game, dropping to the ground and making a small bounce. Press R in the game to reload and see it again.


![end of step 5](/assets/images/physics.png)
