# 3D Tutorial - Step 8 - Ground contact
by Monstrous Software


# Step 8 - Ground contact


## Ray casting for ground contact

So far, in the player controller we are not testing if the player is on the ground.  For example, the player should only be allowed to jump if they are stood on something, not if they are already jumping.
To test if the player is stood on something, we will use a ray cast to check for collisions with geometry just below the player's capsule.

Create the following class to implement ray casting.  It creates a ray which points downwards from the player position. It should be slightly longer than half the player's height, so that it sticks out a bit from 
under the character's feet.  Then we use ODE's spaceCollide2 method to test for collision between the ray and all geoms defined in the world space.  
This uses a callback method to check each pair of potential colliding geoms.  In the callback method, we return the ground normal of there is a collision.  This should normally be pointing upwards if we're on 
a flat surface, but can have another value when we're stood on a slope.  If there is no ground collision, the ground normal is set to zero and the method `isGrounded` returns false.


```java
        public class PhysicsRayCaster implements Disposable {
        
            private final DSpace space;       // reference only
            private final DRay groundRay;
            private GameObject player;
        
            public PhysicsRayCaster(DSpace space) {
                this.space = space;
                groundRay = OdeHelper.createRay(1);        // length gets overwritten when ray is used
            }
        
            public boolean isGrounded(GameObject player, Vector3 playerPos, float rayLength, Vector3 groundNormal ) {
                this.player = player;
                groundRay.setLength(rayLength);
                groundRay.set(playerPos.x, playerPos.z, playerPos.y, 0, 0, -1); // swap Y & Z, point ray downwards 
                groundRay.setFirstContact(true);
                groundRay.setBackfaceCull(true);
        
                groundNormal.set(0,0,0);    // set to invalid value
                OdeHelper.spaceCollide2(space, groundRay, groundNormal, callback);
                return !groundNormal.isZero();
            }
        
            private final DGeom.DNearCallback callback = new DGeom.DNearCallback() {
        
                @Override
                public void call(Object data, DGeom o1, DGeom o2) {
                    GameObject go;
        
                    final int N = 1;
                    DContactBuffer contacts = new DContactBuffer(N);
                    int n = OdeHelper.collide (o1,o2,N,contacts.getGeomBuffer());
                    if (n > 0) {
                        if(o2 instanceof DRay )
                            go = (GameObject)o1.getData();
                        else
                            go = (GameObject)o2.getData();
                        if(go == player)      // ignore collision with player itself
                            return;
                        DVector3 normal = contacts.get(0).getContactGeom().normal;
                        ((Vector3)data).set((float)normal.get(0), (float)normal.get(2), -(float)normal.get(1));	// swap Y&Z
                    }
                }
            };
        
        
            @Override
            public void dispose() {
                groundRay.destroy();
            }
        }   
```



Now let's make use of this new method in the player controller.  We call the ray caster to check if we're on the ground.  If we are we check if we happen to be stood on a slope.
For this we calculate the dot product of the ground normal and the up vector.  The dot product gives us the cosine of the angle.  If the angle is very small, i.e. the surface under the player's feet is 
almost horizontal, then the cosine will be close to 1.  If the cosine is less than one, it means we are standing on a slope.  To prevent the player capsule sliding down the slope, we temporarily
disable gravity for the player.  This trick is explained in more detail in James T. Khan's video tutorial on [dynamic character controllers](https://www.youtube.com/watch?v=O0Deshj2-KU&list=PLjUR2MkQ0cuEda0_f-CoAZBowEN8gNSfJ) (note he is using the Bullet physics library instead).



```
        public void update (GameObject player, float deltaTime ) {
            ...
    
            boolean isOnGround = rayCaster.isGrounded(player, player.getPosition(), Settings.groundRayLength, groundNormal);
            // disable gravity if player is on a slope
            if(isOnGround) {
                float dot = groundNormal.dot(Vector3.Y);
                player.body.geom.getBody().setGravityMode(dot >= 0.99f);
            } else 
                player.body.geom.getBody().setGravityMode(true);
```

The ray length needs to be chosen to stick out a little below the player's capsule. A reasonable value is defined in the Settings class, the value depends on the height of the character:

```
        static public float groundRayLength = 1.2f;
```

Likewise, we can allow jumping only when the character is stood on something.

```
        if (isOnGround && keys.containsKey(jumpKey) )
            linearForce.y =  Settings.jumpForce;
```



This concludes step 8. 
