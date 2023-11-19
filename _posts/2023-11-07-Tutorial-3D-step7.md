# 3D Tutorial - Step 7 - Mesh collisions
by Monstrous Software


# Step 7 - Mesh collisions

## CollisionShapeType MESH

Up to now, we only support basic shapes for collision detection such as boxes, spheres, cylinders and capsules. What if we need a more complex shape?
It is time to add a generic mesh shape where we will construct the collision geometry from the mesh of the ModelInstance.

We're going to use a new GLTF file for this section and while we're at it let's use a settings variable for the file name. 

Add the following to Settings:

    static public final String GLTF_FILE = "models/step12.gltf";

Change `GameScreen.show()` to use this new settings field:

    @Override
    public void show() {

        world = new World(Settings.GLTF_FILE);
        ...
    }


And add the following line to Populator to load in a new game object in the shape of an arch. This will be the first object to use a MESH collision shape which we defined as a type but didn't support yet:

        world.spawnObject(true,"arch",  CollisionShapeType.MESH, false, Vector3.Zero, 1f);

To support this new collision shape type we need to do two things: we need to add code to the PhysicsBodyFactory to create a collision geometry instance (a geom) and we need to add code to create a Model to visualize the shape in debug mode.  The latter part is not strictly necessary because it is only used during development, but it will probably save you from pulling your hair out.

Let's start with the last part and add a case for MESH in the model builder code of createBody():

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
            case MESH:                                          // <----- new
                buildLineMesh(meshBuilder, collisionInstance);  // <----- new
                break;                                          // <----- new
        }

In the first switch in `createBody()` add a place-holder for the MESH case to create a box shaped geom so that at least it will not throw an exception.

        switch(shapeType) {
            case BOX:
                geom = OdeHelper.createBox(physicsWorld.space, w, d, h);    // swap d & h
                break;
            ....
            case MESH:
                geom = OdeHelper.createBox(physicsWorld.space, w, d, h);    // TEMPORARILY
                break;
            default:
                throw new RuntimeException("Unknown shape type");
        }


Now let us create a method in PhysicsBodyFactory to construct a wire frame mesh using LibGDX's MeshPartBuilder:

    // create a wire frame mesh of the collision model instance
    private void buildLineMesh(MeshPartBuilder meshBuilder, ModelInstance instance) {
        Mesh mesh = instance.nodes.first().parts.first().meshPart.mesh;

        int numVertices = mesh.getNumVertices();
        int numIndices = mesh.getNumIndices();
        int stride = mesh.getVertexSize()/4;        // floats per vertex in mesh, e.g. for position, normal, textureCoordinate, etc.

        float[] origVertices = new float[numVertices*stride];
        short[] origIndices = new short[numIndices];
        // find offset of position floats per vertex, they are not necessarily the first 3 floats
        int posOffset = mesh.getVertexAttributes().findByUsage(VertexAttributes.Usage.Position).offset / 4;

        mesh.getVertices(origVertices);
        mesh.getIndices(origIndices);

        meshBuilder.ensureVertices(numVertices);
        for(int v = 0; v < numVertices; v++) {
            float x = origVertices[stride*v+posOffset];
            float y = origVertices[stride*v+1+posOffset];
            float z = origVertices[stride*v+2+posOffset];
            meshBuilder.vertex(x, y, z);
        }
        meshBuilder.ensureTriangleIndices(numIndices/3);
        for(int i = 0; i < numIndices; i+=3) {
            meshBuilder.triangle(origIndices[i], origIndices[i+1], origIndices[i+2]);
        }
    }

All this method does is create triangles corresponding to the original mesh of the ModelInstance. Because meshBuilder was created using 
a GL20.GL_LINES parameter value (instead of GL_TRIANGLES) the new mesh created from these triangles will be created as a wireframe to be used
as debug shape visualization.

![arch](/assets/images/arch.png)

At this point the debug wire frame is misleading because the actual collision shape for the arch is a block.  We can test this by trying to walk through the arch or to shoot balls at it. You cannot go through the arch.

We need to construct a geom that follows the ModelInstance shape. ODE has a method to create a geom from what is calls a TriMesh (triangle mesh).
Modify the MESH case of the first switch statement to the following to create an ODE geom using TriMesh data (i.e. the data for a triangle mesh, which is an array of vertices and an array of indices).

            case MESH:
                // create a TriMesh from the provided modelInstance
                DTriMeshData triData = OdeHelper.createTriMeshData();
                fillTriData(triData, collisionInstance);
                geom = OdeHelper.createTriMesh(physicsWorld.space, triData, null, null, null);
                break;

Then add the following method which will convert a LibGDX mesh to ODE TriMesh data  The logic is very similar to the previous method to construct a wire frame.
Note that in this method we convert to ODE reference frame by swapping Y and Z values per vertex.  Because this messes up the winding order of the triangles, we
also reorder the indices of each triangle to reverse the winding. If we don't do that the collision mesh will be inside out; we can enter the collision geom, but we cannot get out. 


    // convert a libGDX mesh to ODE TriMeshData
    private void fillTriData(DTriMeshData triData, ModelInstance instance ) {
        Mesh mesh = instance.nodes.first().parts.first().meshPart.mesh;

        int numVertices = mesh.getNumVertices();
        int numIndices = mesh.getNumIndices();
        int stride = mesh.getVertexSize()/4;        // floats per vertex in mesh, e.g. for position, normal, textureCoordinate, etc.

        float[] origVertices = new float[numVertices*stride];
        short[] origIndices = new short[numIndices];
        // find offset of position floats per vertex, they are not necessarily the first 3 floats
        int posOffset = mesh.getVertexAttributes().findByUsage(VertexAttributes.Usage.Position).offset / 4;

        mesh.getVertices(origVertices);
        mesh.getIndices(origIndices);

        // data for the trimesh
        float[] vertices = new float[3*numVertices];
        int[] indices = new int[numIndices];

        for(int v = 0; v < numVertices; v++) {
            vertices[3*v] = origVertices[stride*v+posOffset];        // X := x
            vertices[3*v+1] = -origVertices[stride*v+2+posOffset];   // Y := -z
            vertices[3*v+2] = origVertices[stride*v+1+posOffset];    // Z := y
        }
        for(int i = 0; i < numIndices; i++)         // convert shorts to ints
            indices[i] = origIndices[i];

        triData.build(vertices, indices);
        triData.preprocess();
    }

With this code in place, we are able to use arbitrary meshes as collision shapes, for those cases when a box or a sphere is just not good enough.



## Collision Proxy

The next feature we want to add is to allow defining a simplified mesh for use as collision shape when a sphere or box is not a close enough match but the model mesh is perhaps quite complex.
This will make the collision testing more efficient. But also it will help to smooth the movement in game to avoid the player getting stuck on ornamental protrusions or to painstakingly negotiate each step of a staircase.

To do this we need to supply two node names: one for the complex object and one for the simplified object.  The complex object is what we'll be rendering and what the user will see.  The simplified 
object is only used for collision detection and will never be seen in-game.

![](/assets/images/collisionProxy.png)

An example is shown in the image above. On the left is a model of a staircase. This model has 132 vertices (which is actually still very low poly). On the right is a simplification of the staircase as a basic slope which uses just 8 vertices in a nice garish colour just to emphasize it will never be seen in-game.
You can imagine the efficiency benefit is more pronounced when we have super detailed models.

To support this we will adapt the spawnObject() method in the World class.  Instead of one, we now pass two node names; one of the node that will be rendered and one of the node that will be used for collision testing (the proxy).  The proxy is optional and can be null.  Typically, you will be using the proxy node in combination with the collision shape type MESH.

Both nodes are loaded as scene and transformed using the position parameter and the node's transform.  If a proxy node was specified, this will be used to construct the PhysicsBody instead of the original node.

```
        public GameObject spawnObject(boolean isStatic, String name, String proxyName, CollisionShapeType shapeType, boolean resetPosition, Vector3 position, float mass){
            Scene scene = loadNode( name, resetPosition, position );
            ModelInstance collisionInstance = scene.modelInstance;
            if(proxyName != null) {
                Scene proxyScene = loadNode( proxyName, resetPosition, position );
                collisionInstance = proxyScene.modelInstance;
            }
            PhysicsBody body = factory.createBody(collisionInstance, shapeType, mass, isStatic);
            GameObject go = new GameObject(scene, body);
            gameObjects.add(go);
            isDirty = true;         // list of game objects has changed
            return go;
        }
    
        private Scene loadNode( String nodeName, boolean resetPosition, Vector3 position ) {
            Scene scene = new Scene(sceneAsset.scene, nodeName);
            if(scene.modelInstance.nodes.size == 0)
                throw new RuntimeException("Cannot find node in GLTF file: " + nodeName);
            applyNodeTransform(resetPosition, scene.modelInstance, scene.modelInstance.nodes.first());         // incorporate nodes' transform into model instance transform
            scene.modelInstance.transform.translate(position);
            return scene;
        }
```

Add the following line to Populator to try this out by loading a staircase and a proxy node.

        world.spawnObject(true, "arch", "archProxy", CollisionShapeType.MESH, false, Vector3.Zero, 1f);

For all the other lines in the `populate()` method, insert a `null` after the node name. Idem for the `spawnObject()` call in the `shootBall()` method.

Below we can see with the in-game debug view how the staircase is approximated by a simple prism shape.

![stairs](/assets/images/staircase.png)


This concludes step 7.
