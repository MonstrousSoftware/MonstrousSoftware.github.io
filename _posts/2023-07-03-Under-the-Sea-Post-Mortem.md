# Post Mortem - Under the Sea

This game was developed for the LibGDX game jame #25 ([1]) which had as theme ‘under water’.  This post looks back at some things encountered during its short development cycle.

![screenshot2](https://github.com/MonstrousSoftware/MonstrousSoftware.github.io/assets/49096535/0f0620ba-9296-4de3-929e-85f9c7c818d7)
>  Figure 1 - Screenshot of the final game.

My objective was to develop a simple game in one week which would run on web browsers, so that nobody needs to download a JAR file.
The idea for the game came from one of Sebastian Lague’s Youtube videos ([2]), namely to generate a terrain using marching cubes and exploring this world with a submarine.

## Marching Cubes algorithm

Marching Cubes are a bit like voxels but with smoother corners.  I could reuse some code I had lying around for a voxel world.

The terrain is generated with a 3D Perlin noise function.  But where for my voxel world each point in the three-dimensional world was either on or off (solid block versus air), we now generate a 3d matrix of density values, scaled from 0 to 255.

The terrain consists of 7 by 7 chunks. In principle this could be extended quite easily to an infinite terrain by creating and disposing chunks on the fly, but for this little game a fixed size world seemed sufficient. 
Each chunk has 31x31x127 cubes which are constructed based on 32x32x128 density sample points. (128 in the vertical direction).

The algorithm takes every density value above 64 to be solid (rock) and values below 64 to be empty (water). Then for each cube in the chunk, it generates triangles to connect cube edges depending on which of the 8 cube vertices are solid or not.  By generating all the triangles for each cube in the chunk we construct the surface of the rock. 

![marching-cubes](https://github.com/MonstrousSoftware/MonstrousSoftware.github.io/assets/49096535/11256320-899d-4a2c-866e-86d5b34107d5)

> Figure 2 - Image by user Ryoshuro from Wikipedia article on Marching Cubes


Figuring out which configuration of vertices result in which set of triangles is done via a large lookup table which I copied from the internet. In the first instance the triangle vertices were all exactly on the mid-point of a cube edge. Later, I modified this to select a point on the edge based on an interpolation of both vertex densities which gives a much more organic mesh.

 ![before-after](https://github.com/MonstrousSoftware/MonstrousSoftware.github.io/assets/49096535/0956bb54-11bf-471f-9da1-8fe11402524a)

>  Figure 3 – Marching Cubes before and after interpolation per edge

## Smooth Shading
To improve the appearance, I wanted to move from a flat shaded look to smooth shading. (To be honest this was done after the jam deadline).  Key to this is to move from one normal vector per triangle to a normal vector per vertex that is averaged over the different triangles the vertex belongs to.
Originally, when creating the mesh, I would calculate the normal vector per triangle and use that value as attribute for each of the vertices. Vertices were not shared between triangles.  
For smooth shading, I first generate the vertices as before and place them into a vector array. Then I use a hash map to combine vertices that are at the same position.  The index table is created at the same time. The normal vectors of combined vertices are averaged (simply add the normal vectors of each combined vertex and then renormalize them at the end).
This has the added benefit of making the memory footprint of the mesh much smaller, by almost a factor of 8. 

To illustrate, the following diagrams show the before and after pictures.  The green arrows are the face normal. The blue arrows are the vertex normal.  In the first picture, there are size vertices and each vertex normal corresponds to the face normal.  In the second picture, there are only four vertices and for v1 and v2 the vertex normal are averaged between the face normal.  The index array defines how the triangles are linked to vertices.

![tris-before-after](https://github.com/MonstrousSoftware/MonstrousSoftware.github.io/assets/49096535/7d0b8743-2a5c-4284-b821-b04cc4da5170)

>  Figure 4 – interpolating normal vectors per vertex

The effect can be seen in the following image.

![smooth](https://github.com/MonstrousSoftware/MonstrousSoftware.github.io/assets/49096535/5a21cc11-ba19-4f3e-a17f-ea7e37c77121)

>  Figure 5 – smooth shaded terrain


To make the terrain a little more interesting, I apply a texture on the triangles. I just use the vertex x and z as texture coordinated u and v, which means the texture aligns nicely on horizontal surfaces.  On the other hand, we can notice some stretching on vertical surfaces.
A technique to improve this is called tri-planar mapping, but there was no time to get this working for the jam.

![textured](https://github.com/MonstrousSoftware/MonstrousSoftware.github.io/assets/49096535/1aad4033-8517-48eb-ae6f-1f1612536b05)

>  Figure 6 – terrain with a repeating texture applied 

## Collision detection

Collision detection was initially done by testing one point at the front tip of the submarine and one point at the rear.  For both points, I checked the density value to see if it is above the threshold value for being solid rock. For this purpose, I keep each chunk’s density matrix around (it could otherwise be discarded after the mesh construction). 
It is simple and fast, because we make use of the underlying data that defines the terrain; we don’t need to test thousands of triangles. Unfortunately, it only catches head-on collisions (or when you reverse into rock). It does not catch when the side of the submarine scrapes the rock.  This could be mitigated by defining more collision points, but this is not the path I chose.

I defined another data structure per chunk called a distance field, which just stores for each cube the distance to the rock surface.  This is derived from the density matrix using a type of flood fill algorithm. Using this distance field, I could quickly test how far the submarine is from the rock surface to spot a collision.  Since the submarine has a capsule shape, we can get a reliable collision detection by testing the centre of the front sphere and the rear sphere.
Unfortunately, it took a long time to calculate the distance field, which means the start-up time of the game became intolerable.  I considered loading the distance field from file, but this would mean another 6MB to download.  I abandoned this approach.

Next, I looked at using the ODE4j physics library for collision detection.  This is derived from the ODE physics library and ported to Java.  Antz recently ported this to LibGDX.  The advantage of the Bullet physics library is that, being pure Java, it is supported by the HTML client, whereas Bullet, being a C++ library wrapped in a JNI layer, is not.  
The ODE library has a number of collision functions between geometry objects. We are only interested in collisions between the submarine and the terrain.

To filter collisions we can define category flags, so that terrain can only collide with the submarine (not with other terrain chunks). 

    public static int CAT_TERRAIN = 1;
    public static int CAT_SUBMARINE = 2;

 For the submarine we use a simple capsule shape:

        subCapsule = OdeHelper.createCapsule(space, 1, 3); // radius of caps, length without caps
        subCapsule.setCategoryBits(CAT_SUBMARINE);
        subCapsule.setCollideBits(CAT_TERRAIN);

For the terrain surface of each chunk we use a triangle mesh:

  DTriMesh triMesh = OdeHelper.createTriMesh(space, chunk.triMeshData, null, null, null);
  triMesh.setCategoryBits(World.CAT_TERRAIN);
  triMesh.setCollideBits(World.CAT_SUBMARINE);

The mesh data for an ODE4j TriMesh is very similar to the LibGDX mesh, but can only three floats per vertex for the position (no normal vectors, UV texture coordinates, color or any other attributes) and the index array must be an array of int rather than an array of short.

Then we can define a callback function that will tell us when there is a collision:

    private DGeom.DNearCallback nearCallback = new DGeom.DNearCallback() {
        @Override
        public void call(Object data, DGeom o1, DGeom o2) {
            nearCallback(data, o1, o2);
        }
    };

    private void nearCallback (Object data, DGeom o1, DGeom o2) {
        final int N = 4;
        DContactBuffer contacts = new DContactBuffer(N);
        int n = OdeHelper.collide (o1,o2,N,contacts.getGeomBuffer());
        if (n > 0) 
            submarine.collide();
    }

## Collision response

Until now, the collision response was very simple: set the submarine velocity to zero and display a large warning message.
Having integrated ODE4j already, it would be interesting to use the physics library also for collision response.  The submarine was modelled as a dynamic rigid body in ODE7j and the terrain as a static kinetic body (i.e. a body of infinite mass).  The call back function was modified to create a contact joint between the colliding bodies, which will push the submarine away from the rock on collision.

    private void nearCallback (Object data, DGeom o1, DGeom o2) {
        final int N = 4;
        DContactBuffer contacts = new DContactBuffer(N);
        int n = OdeHelper.collide (o1,o2,N,contacts.getGeomBuffer());
        if (n > 0) {
            submarine.collide();
            for (int i=0; i<n; i++) {
                DContact contact = contacts.get(i);
                DJoint c = OdeHelper.createContactJoint(dworld,contactgroup,contact);
                c.attach (o1.getBody(), o2.getBody());
            }
        }
    }

I experimented with using a more physics-based approach to the submarine movement, e.g. that moving the rudder or the moving the hydroplanes exerts a torque on the submarine’s rigid body.  The result was may be a bit more realistic, but from a game play point of view it made the submarine very hard to control. It ended up doing rolls and loopings.  So in the end I settled for having the rudder and hydroplane positions affect the submarine orientation directly.

## GDX-GLTF
I use the gdx-gltf library to import the submarine model (and the easter egg) in gltf format exported directly from Blender.  Importing like this is easier than using the fbx format and using the standalone fbx-convert tool 
and maintains a higher fidelity.  This library provides a better shader than the default LibGDX one which makes the models look much nicer, for example for metallic surfaces.

## TeaVM
For the web client, I used the Gdx-teaVM library rather than the standard HTML/GWT library.  It is faster to compile and it gives better performance in the browser. 

[1]: https://www.youtube.com/watch?v=em0cy5iPmpg&t=5548s&ab_channel=Raeleus "LibGDX Jam June 2023 Review"

[2]: https://www.youtube.com/watch?v=M3iI2l0ltbE&ab_channel=SebastianLague "Sebastian Lague coding adventures"
