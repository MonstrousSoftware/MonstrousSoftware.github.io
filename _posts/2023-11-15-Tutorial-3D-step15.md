# 3D Tutorial - Step 15 - Unifying the Reference Frame
by Monstrous Software


# Step 15 - Unifying the Reference Frame

After doing a bit more research (which I should have done earlier of course), it turns out we don't need to use different coordinate systems between
the graphics code and the physics code as we introduced in step 5.

We discussed before that the physics library ODE is agnostic with regard to the coordinate systems that you want to use.  The reason we perform coordinate conversion 
between the two parts of the program is that collision cylinders and capsules in ODE are always created along the Z axis and for our characters we want the collision capsules
to be upright.

There is however a trick in ODE to define an offset rotation between the rigid body and the corresponding geom. This allows us to rotate the cylinder and capsule
geoms immediately after creation so that they are aligned along the Y axis. This is an extra rotation that is always applied on top of any rotation that is applied 
to the rigid body.  This offset rotation is set with `geom.setOffsetQuaternion()`.  

If we apply this, we can use "Y is up" also in the physics code and the conversion between ODE vectors and LibGDX vectors become much more straightforward, and we don't have 
to swap Z and -Y coordinates anymore (with lots of chance for errors).

To change the ODE coordinate system, go to `PhysicsWorld.reset` and change the following lines 
```java
       space = OdeHelper.createSapSpace( null, DSapSpace.AXES.XYZ );           
       world.setGravity (0, 0, Settings.gravity); 
```
to the following:
```java
        space = OdeHelper.createSapSpace( null, DSapSpace.AXES.XZY );
        world.setGravity (0,  Settings.gravity, 0); 
```
From now on, ODE uses the same coordinate system as LibGDX and OpenGL, i.e. Y is up.


To make the collision shapes also be aligned to the up axis, add the following code to `PhysicsBody.createBody`. In case the geometry shape is a cylinder or capsule, create a quaternion
to define a 90-degree rotation around the X axis.  This will change the Z alignment to a Y alignment. For the other geometry shapes such as sphere or box, this is not needed because they are symmetrical.
```java
            if(shapeType == CollisionShapeType.CYLINDER || shapeType == CollisionShapeType.CAPSULE) {
                // rotate geom 90 degrees around X because ODE geom cylinders and capsules shapes are created using Z as long axis
                // and we want the shape to be oriented along the Y axis which is up.
                DQuaternion Q = DQuaternion.fromEulerDegrees(90, 0, 0);     // rotate 90 degrees around X
                geom.setOffsetQuaternion(Q);    // set standard rotation from rigid body to geom
            }
```

Then everywhere we convert between ODE vectors and LibGDX vectors (mostly in PhysicsBody, PhysicsBodyFactory and PhysicsRayCaster) 
we can use the same order of x, y and z and we don't have to add minus signs anywhere. For example, in the PhysicsBody class:
```java
            public Vector3 getPosition() {
                DVector3C pos = geom.getPosition();
                position.x = (float) pos.get0();
                position.y = (float) pos.get1();
                position.z = (float) pos.get2();
                return position;
            }
        
            public void setPosition( Vector3 pos ) {
                geom.setPosition(pos.x, pos.y, pos.z);
                // if the geom is attached to a rigid body it's position will also be changed
            }
```
Note that the offset rotation of the geom means that the geom may have a different orientation than its corresponding rigid body.
For the debug viewer, we want access to the rigid body orientation.  We add a new method for this.
```java
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
```
Of course in `PhyicsWorld.reset()` we now also need to define gravity to be down the Y axis instead of the Z axis.
```java
        world.setGravity (0,  Settings.gravity, 0);
```

## Refactoring of debug object view

While we're doing some cleanup, we also notice that `PhysicsBody` has a render method, which should more properly be part of `PhysicsView` so that
rendering and physics is better separated.  For example, the colours that are used to show active or sleeping bodies are more at home in the PhysicsView class.
The `render` method is therefore rewritten as follows, and we get rid of `PhysicsBody.render()`:

```java   
        // colours to use for active vs. sleeping geoms
        static private final Color COLOR_ACTIVE = Color.GREEN;
        static private final Color COLOR_SLEEPING = Color.TEAL;
        static private final Color COLOR_STATIC = Color.GRAY;
        
        public void render( Camera cam ) {
            modelBatch.begin(cam);
            int num = world.getNumGameObjects();
            for(int i = 0; i < num; i++) {
                GameObject go = world.getGameObject(i);
                if (go.visible)
                    renderCollisionShape(go.body);
            }
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
```

## Moment of inertia

In the physics code, we did not consider so far how the mass of a body is distributed over the shape of a body.  This is also called the 'moment of inertia'.
ODE allows to define a shape for the distribution of mass, which can give a more realistic behaviour for example when a force is applied to the edge of an object.
(To be fair, the difference is rather subtle, and this change is entirely optional).
For example, for a spherical mass we use `massInfo.setSphere(1, radius)` where 1 is arbitrarily used as mass density.  
If we assume all game objects have the same mass density, we can let the 
mass be automatically calculated from the shape and its size.  This means that larger objects will be heavier than smaller ones.
We now no longer have to define a mass for every object we create.  Note that we will need to tweak the force values as this changes the mass of the various objects.

Here is the new version of `createBody` using mass derived from the shape and the offset rotation we discussed earlier:

```java
    public PhysicsBody createBody( ModelInstance collisionInstance, CollisionShapeType shapeType, boolean isStatic) {
        BoundingBox bbox = new BoundingBox();
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
                massInfo.setBox(1, w, h, d);
                break;
            case SPHERE:
                diameter = Math.max(Math.max(w, d), h);
                radius = diameter/2f;
                geom = OdeHelper.createSphere(physicsWorld.space, radius);
                massInfo.setSphere(1, radius);
                break;
            case CAPSULE:
                diameter = Math.max(w, d);
                radius = diameter/2f; // radius of the cap
                len = h - 2*radius;     // height of the cylinder between the two end caps
                geom = OdeHelper.createCapsule(physicsWorld.space, radius, len);
                massInfo.setCapsule(1, 2, radius, len);
                break;
            case CYLINDER:
                diameter = Math.max(w, d);
                radius = diameter/2f; // radius of the cap
                len = h;     // height of the cylinder between the two end caps
                geom = OdeHelper.createCylinder(physicsWorld.space, radius, len);
                massInfo.setCylinder(1, 2, radius, len);
                break;
            case MESH:
                // create a TriMesh from the provided modelInstance
                DTriMeshData triData = OdeHelper.createTriMeshData();
                fillTriData(triData, collisionInstance);
                geom = OdeHelper.createTriMesh(physicsWorld.space, triData, null, null, null);
                massInfo.setBox(1, w, h, d);
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
                DQuaternion Q = DQuaternion.fromEulerDegrees(90, 0, 0);     // rotate 90 degrees around X
                geom.setOffsetQuaternion(Q);    // set standard rotation from rigid body to geom
            }
        }
        ...
    }
```

This concludes step 15.
