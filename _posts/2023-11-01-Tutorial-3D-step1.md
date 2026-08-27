# Tutorial on creating a 3D game with LibGDX - Intro
by Monstrous Software (updated August 2026)

## Introduction
In this tutorial series we are going to build a basic first-person shooter using LibGDX.

This will be a 3d single player game. 
We will be targeting the desktop version and a web version using the gdx-teavm extension.
We will use the extension gdx-gltf to load 3d assets using the GLTF file format, and we'll see how we can export assets from modeling software such as Blender.
We will add physics using the ODE4j library.

This will not be a polished game, you can try it out on [itch.io](https://monstrous-software.itch.io/fps-demo) now if you want.
The idea is to show how you can develop a 3d game using LibGDX and some of the popular extensions.

The development is shown as a number of steps, where every step we add something new (and sometimes we remove something old).
There is a [github repository](https://github.com/MonstrousSoftware/Tut3D) which provides all the code corresponding to this tutorial and every step of this 
tutorial is provided as a release.  Using the release snapshots you can step through the history of the code. Each step results in 
a runnable version of the app.

## Index
- [step 1: project setup, gdx-liftoff, 3d scene](https://monstroussoftware.github.io/2023/11/01/Tutorial-3D-step1.html)
- [step 2: FPS camera controller](https://monstroussoftware.github.io/2023/11/02/Tutorial-3D-step2.html)
- [step 3: gdx-gltf, importing from Blender](https://monstroussoftware.github.io/2023/11/03/Tutorial-3D-step3.html)
- [step 4: multiple objects, grid view](https://monstroussoftware.github.io/2023/11/04/Tutorial-3D-step4.html)
- [step 5: physics, collisions, rigid bodies](https://monstroussoftware.github.io/2023/11/05/Tutorial-3D-step5.html)
- [step 6: spawning bullets, debug view, player/camera controller](https://monstroussoftware.github.io/2023/11/06/Tutorial-3D-step6.html)
- [step 7: mesh collision shapes, collision proxy objects](https://monstroussoftware.github.io/2023/11/07/Tutorial-3D-step7.html)
- [step 8: ray casting, ground collision](https://monstroussoftware.github.io/2023/11/08/Tutorial-3D-step8.html)
- [step 9: game object types, pickups](https://monstroussoftware.github.io/2023/11/09/Tutorial-3D-step9.html)
- [step 10: enemy characters, bullets, health](https://monstroussoftware.github.io/2023/11/10/Tutorial-3D-step10.html)
- [step 11: asset manager, sounds](https://monstroussoftware.github.io/2023/11/11/Tutorial-3D-step11.html)
- [step 12: GUI](https://monstroussoftware.github.io/2023/11/12/Tutorial-3D-step12.html)
- [step 13: gun](https://monstroussoftware.github.io/2023/11/13/Tutorial-3D-step13.html)
- [step 14: scope view](https://monstroussoftware.github.io/2023/11/14/Tutorial-3D-step14.html)
- [step 15: unified reference frame](https://monstroussoftware.github.io/2023/11/15/Tutorial-3D-step15.html)
- [step 16: game controller support](https://monstroussoftware.github.io/2023/11/16/Tutorial-3D-step16.html)
- [step 17: head bobbing](https://monstroussoftware.github.io/2023/11/17/Tutorial-3D-step17.html)
- [step 18: full screen mode](https://monstroussoftware.github.io/2023/11/18/Tutorial-3D-step18.html)
- [step 19: navigation mesh](https://monstroussoftware.github.io/2023/11/19/Tutorial-3D-step19.html)
- [step 20: framerate independence](https://monstroussoftware.github.io/2023/11/20/Tutorial-3D-step20.html)
- [step 21: navigation mesh generation](https://monstroussoftware.github.io/2023/11/21/Tutorial-3D-step21.html)


## Prerequisites
You'll need an IDE (integrated development environment). I would recommend [IntelliJ IDEA](https://www.jetbrains.com/idea/) by JetBrains. The community version is free and is more than sufficient for this project.

To view or edit the 3d models, you can use Blender which is free to download from [Blender.org](https://blender.org).

## Step 1 - Lift off

To build the initial framework of our project we'll use the tool `gdx-liftoff`.  This is a replacement for the classic `gdx-setup` tool.

Download the latest version of gdx-liftoff from here: [https://github.com/libgdx/gdx-liftoff](https://github.com/libgdx/gdx-liftoff).
It comes in the form of a jar file with a name like `gdx-liftoff-VERSION.jar`.

Double-click on the jar file. If nothing happens, go to the command line and type: 

```    java -jar gdx-liftoff-VERSION.jar``` (substitute the version you downloaded).

A window pops up, asking for details of the project you want to build. There are a number of screens. We'll go through them one by one and then gdx-liftoff
will generate the project outline for us.

In the main screen we fill in:
- the name of the project: 'Tut3D'
- a package identifier, for example 'com.yourcompany.yourgame'
- the name of the main class. I tend to just call it 'Main' for ease of reference.

On the next screen, for the platforms, we select 'Desktop'. If you'd like to try building a web version, select 'HTML (TeaVM)'.  
The option 'Core' always needs to be selected.

Web versions are great for game jams or to reach a large audience, because people can play your game without having to download files.
The web versions have some limitations though, so for this tutorial we'll mainly focus on the desktop version.

![lift-off screen shot](/assets/images/liftoff-new1.png)

Under the tab 'Extensions', select 'Controllers' for controller support (we will get back to that in Step 16).

Under 'Templates' select 'Game'.  This will make it easy to switch between different screens in our game, for example a menu screen and a game screen.

Go to the 'Third party' tab and select 'ControllerMapping' and 'ControllerScene2D' for more controller support.
Scroll down and select 'gdx-gltf' which we'll use to import 3d models and for Physics Based Rendering.

On the last tab, select the latest LibGDX version. It is recommended to choose Java version 21.
Select 'Add GUI assets' in case we'll need some GUI elements and 'Add README'.

Fill in the directory for the project.  You can either create an empty folder first and then navigate to it using the 'Browse' icon, or type in a name for the folder and press the 'Create Folder' button.

Press the button 'Generate'. The tool will now create a project directory.

Press the button to open the project in IntelliJ IDEA or navigate to the peoject folder and double-click on build.gradle. 
Or alternatively, open IntelliJ IDEA and use File/Open to open build.gradle.
If IDEA asks to open build.gradle as file or as project, answer 'as project'.

When we view the project directory from IDEA it should look as follows:

![project directory](/assets/images/projdir2.png)

Once IDEA has opened the project it will automatically start running some Gradle tasks and download the necessary libraries. 
This will take a little while.

Open the Gradle menu on the right hand side (the elephant icon). Navigate to lwjgl3/Tasks/application and double-click on the run icon.
(Note: the module lwjgl3 is commonly known as the desktop version).
This will compile the project and then open a window called 'Tut3D' with a slightly disappointing black screen for content.

At this point the project structure is created, and we need to start filling in some content.

## Our first 3D scene: Hello Cube

In the Project window, open the `core` module and drill down until we see the Java classes. Rename the class FirstScreen.java to GameScreen.java.  This screen will be the most important one in the game because it is where the game is played. Later we can add screens like a main menu, a splash screen, an options screen, etcetera.

To quickly rename a class in the IDE, right click FirstScreen.java in the Project view. Select Rename (or press Shift F6) and then rename.
This will rename the source file, the name of the class file and also the reference to the class in Main.java.

For a very basic 3D scene, we need at minimum a Camera, a ModelInstance and a ModelBatch to render the model instance following the
camera settings. Let's make an application which shows a single cube.

In the GameScreen class, let's add some fields: a camera, a model, a ModelInstance and a ModelBatch.

```java
    private PerspectiveCamera cam;
    private Model cubeModel;
    private ModelInstance cubeInstance;
    private ModelBatch modelBatch;
```

When you add these lines to the IDE, the class names may appear in red because the class is not known.
Right-click on the class name and select import and the IDE will generate a corresponding import statement.

We will create the perspective camera in the `show` method.  It is this camera that gives us a 3d view of the world.
The first parameter is the camera's viewing angle in degrees, you can choose a different value if you want more of a fish eye view.
Experiment with what feels good for your game. We set the position of the camera at x=10, y=1.5 and z=5. In the LibGDX coordinate system the Y dimension is the up dimension and 1.5 means our eyes are 1.5 meters above ground level if we choose our units to correspond to meters.
The camera is set to look at the world origin, i.e. the point at x=0, y=0, z=0.
Then we set the near and far clipping plane. This determines the depth range of the viewing frustum.
Anything closer to the camera than the near plane or further that the far plane will not be shown.
Always set the far value large enough to see all of your scene. Setting it too large however is detrimental for the depth resolution and can lead to Z-fighting artifacts.
After changing camera parameters it is important to call update() so that they take effect.

```java
    @Override
    public void show() {
        // Prepare your screen here.

        cam = new PerspectiveCamera(67, Gdx.graphics.getWidth(),  Gdx.graphics.getHeight());
        cam.position.set(10f, 1.5f, 5f);
        cam.lookAt(0,0,0);
        cam.near = 1f;
        cam.far = 300f;
        cam.update();
    }
```

To create a model batch, we just have to call its constructor. Add the following line to the `show` method:

```java
        modelBatch = new ModelBatch();
```

And dispose of the model batch in the `dispose` method:

```java
    @Override
    public void dispose() {
        modelBatch.dispose();
    }
```

A ModelInstance is a 3d object in your game world. It is created from a Model which defines the appearance of the object (the mesh and materials).
A Model acts as a template and a ModelInstance is what is actually rendered in your game. For example, you make have a model of a car and then have three cars driving around in the game (the model instances).  

A ModelInstance has what's known as a transform matrix. This is a 4 by 4 matrix that defines the position, orientation and scale of the instance.
For 3D rendering, 4 by 4 matrices are very convenient and used a lot. When we move or rotate an instance we do this by changing the transform matrix of the instance.
(The position defined by a transform matrix is also known as the translation).

Simple geometric models can be created in code using the ModelBuilder.  
For more complicated models, we will load them from a resource file (see Step 3).

Add the following code to the `show` method. This creates a box model of size 5 by 5 by 5.
The model will have a green material.
The diffuse color is what you'd normally think of as the surface color.
A surface can also have a specular color for highlights or an emissive color if it emits light.
The mesh will be constructed using a position vector and a normal vector for each vertex. These are the vertex attributes.
The position attribute is mandatory for any mesh.  The normal vector is used for lighting,
in particular to determine the angle between a surface and a light source.  
A normal vector is a unit vector (i.e. with a length of one) in the direction that the surface is facing.
The vectors are filled in automatically by the utility method ```createBox```.
Once we have created the model, we create an instance at the origin, i.e. position x = 0, y = 0, z = 0.


```java
        ModelBuilder modelBuilder = new ModelBuilder();
    
        // create model
        cubeModel = modelBuilder.createBox(5f, 5f, 5f,
                new Material(ColorAttribute.createDiffuse(Color.GREEN)),
                VertexAttributes.Usage.Position | VertexAttributes.Usage.Normal);
    
        // create model instance
        cubeInstance = new ModelInstance(cubeModel, 0, 0, 0);
```

The model will have to be disposed when we exit the application to release its resources. Add this to the `dispose` method:
A model instance does not need to be disposed. If you're not sure what needs to be disposed, check if the object class has a dispose() method.
For example, you need to dispose each Model, but not a ModelInstance.
You need to dispose ModelBatch, but not the PerspectiveCamera.

```java
    @Override
    public void dispose() {
        modelBatch.dispose();
        cubeModel.dispose();
    }
```

An illustration of the perspective camera viewing the cube may clarify what we have set up now (the camera 'far' distance is not shown to scale):
![scene overview](/assets/images/camview.png)


Now that we have a camera, a model batch and a model instance, we can render the scene.  
The `render` method of `GameScreen` will be called for each frame, typically 60 times per second.
We use it to render the screen contents, but we also use it to update whatever is happening in the game. 

To render, we clear the screen to the background color and clear the depth buffer using `ScreenUtils.clear()`.
And then we use the model batch to render one or more instances.

Add the following code to the `render` method:

```java
    @Override
    public void render(float delta) {
        // Draw your screen here. "delta" is the time since last render in seconds.
        ScreenUtils.clear(Color.TEAL, true);
        modelBatch.begin(cam);
        modelBatch.render(cubeInstance);
        modelBatch.end();
    }
```

Running your program should now result in a window like this:

![hello cube](/assets/images/helloCube.png)


To show an example of updating the scene and how you can modify an instance's transform, add the following
line to the start of the `render` method:

```java
        cubeInstance.transform.rotate(Vector3.Y, 45f * delta);
```
This rotates the model instance around the Y axis by 45 degrees per second.

If we resize the window, the cube may get stretched and that is not what we want.  Add the following lines to the `resize` method to
make sure the scene is rendered in the correct proportions:

```java
    @Override
    public void resize(int width, int height) {
        // If the window is minimized on a desktop (LWJGL3) platform, width and height are 0, which causes problems.
        // In that case, we don't resize anything, and wait for the window to be a normal size before updating.
        if(width <= 0 || height <= 0) return;

        // Resize your screen here. The parameters represent the new window size.
        cam.viewportWidth = width;
        cam.viewportHeight = height;
        cam.update();
    }
```

Then we still have a few empty methods left: `pause`, `resume` and `hide`. Instead of that we can
change the class definition from
```java
    public class GameScreen implements Screen 
```
 to 
```java
    public class GameScreen extends ScreenAdapter 
``` 
 and get rid of those methods.
        
If you want to compare the code we've written so far you can look at release 'step-0.5' in the GitHub repository.

## Beyond the Cube 

Now that we can render a simple cube, let's add some more to the code.

We will add a camera controller so that we can move the camera around, we'll improve the lighting by defining an environment,
we'll show how to define a textured model.

### Lighting

The environment is defined to add some lighting: some ambient light and a directional light.
Ambient light is light that just appears everywhere. It's used as filler light to prevent the lighting being too harsh.
Directional light is light that comes from a particular direction, such as the sun. For light that comes from above, the direction vector has to point downwards, i.e. with y < 0.
You can only have a limited number of directional lights for shader performance reasons, by default 2, but you can increase that through shader configuration.
Light colors are defined as red, green and blue values. The direction vector as X, Y, Z.

Add a class member called environment and add the following lines to the `show` method:

```java
    private Environment environment;
```

```java
        environment = new Environment();
        environment.set(new ColorAttribute(ColorAttribute.AmbientLight, 0.2f, 0.2f, 0.2f, 1f));
        environment.add(new DirectionalLight().setColor(0.5f, 0.5f, 0.5f, 1.0f).setDirection(-0.3f, -0.8f, -0.2f));
```

This environment needs to be passed as an additional parameter to `ModelBatch#render` in the `render` method:

```java
        modelBatch.render(cubeInstance, environment);
```

With better lighting, it is easier to recognize the shape of objects:

![hello lit cube](/assets/images/helloCubeLit.png)

### Camera controller

In the GameScreen class, let's add some fields: a camera, a camera controller, an environment, a model, a texture, an array of ModelInstance and a ModelBatch.

```java
    private PerspectiveCamera cam;
    private CameraInputController camController;
    private Environment environment;
    private Model modelGround;
    private Texture textureGround;
    private Array<ModelInstance> instances;
    private ModelBatch modelBatch;
```


Next we will set up a camera controller that will allow us to move the camera around.  
We'll use a standard one from LibGDX, the CameraInputController which lets you orbit
the camera around a central point using the mouse. We'll replace this later with other
camera controllers. The camera controller is an input processor, so we tell GDX to
send input events (e.g. mouse movements) to this camera controller.

Add a new class member and create the camera controller in the `show` method, after the camera is created.

```java
    private CameraInputController camController;
```

```java
       camController = new CameraInputController(cam);
       Gdx.input.setInputProcessor(camController);
```

Then call the camera controller update method at the start of the `render` method.  This will update the camera following mouse movements 
always orbiting the camera around the origin. The scroll wheel can be used to zoom in and out.   

```java
        camController.update();
```

### Textured ground surface

We'll define a box to act as ground level.  Let's load an image to be used as a texture.
Create or download a ground texture, e.g. the one shown below created by Katsukagi which is available at [3dtextures.me](https://3dtextures.me/2022/05/21/stylized-stone-floor-005/) and place it in the folder `assets/textures`
Replace the texture filename in the code as needed.
We use a TextureRegion to tile the texture ten times across the model, and we enable mip mapping to make the texture look good from different distances.

```java
        textureGround = new Texture(Gdx.files.internal("textures/Stylized_Stone_Floor_005_basecolor.jpg"), true);
        textureGround.setFilter(Texture.TextureFilter.MipMapLinearLinear, Texture.TextureFilter.Linear);
        textureGround.setWrap(Texture.TextureWrap.Repeat, Texture.TextureWrap.Repeat);
        TextureRegion textureRegion = new TextureRegion(textureGround);
        int repeats = 10;
        textureRegion.setRegion(0,0,textureGround.getWidth()*repeats, textureGround.getHeight()*repeats );
```

Now we'll use ModelBuilder to create a model for the ground.
The model is just a simple box measuring 100 by 100 in width and depth and 1 in height.
We apply the texture that we just loaded as the diffuse texture.
Then we indicate that for each vertex of the model, we'll need a position, 
a normal vector which is important for lighting and the texture coordinates because we are applying a texture on the model.

```java
        ModelBuilder modelBuilder = new ModelBuilder();

        // create model
        groundModel = modelBuilder.createBox(100f, 1f, 100f,
            new Material(TextureAttribute.createDiffuse(textureRegion)),
            VertexAttributes.Usage.Position | VertexAttributes.Usage.Normal | VertexAttributes.Usage.TextureCoordinates);
```

This model needs to be disposed in the `dispose` method.
```java
        groundModel.dispose();
```

Now that we have a model we can create a ModelInstance from it.  This is an instantiation of the
model in a particular position. In this case we will only make one model instance 
and place it at (0,-1,0) so that the top of the box is exactly at Y=0 which we'll use as a 
convenient ground level.  Meantime, we also have to change the position of `cubeInstance` to be at (0f, 2.5f, 0f) so that
it will rest on the ground level.



It is handy to use an array of ModelInstance in case we want to add more instances later. Add the following class member: 
```java
    private Array<ModelInstance> instances;
```

Then create the array in the `show` method and add the model instance to it. Also add the cube instance to it.

```java
        // create and position model instances

        instances = new Array<>();
        instances.add(new ModelInstance(groundModel, 0, -1, 0));	// 'table top' surface
        instances.add(cubeInstance);
```

The array of model instances can be rendered very easily by the model batch by just passing the whole array:

```java
        modelBatch.begin(cam);
        modelBatch.render(instances, environment);
        modelBatch.end();
```

After all this code, we should now have a very basic 3d scene: we appear to be standing on a texture box that is floating in space.
You can use the mouse to change the view (hold down left or right mouse button) and the WASD keys to move around, albeit in a rather clumsy manner. 
You can use the mouse wheel to zoom.


![step1.png](/assets/images/step1.png)


## Desktop launcher

Add the following lines to the desktop launcher (Lwjgl3Launcher) to increase the size of the window and to activate anti-aliasing to reduce the jagged lines where the ground box meets the sky.
Increase the size of the window to a size you're comfortable with, it depends a bit on what monitor you are using.

```java
        configuration.setWindowedMode(1280, 720);
        configuration.setBackBufferConfig(8, 8, 8, 8, 16, 0, 4);
```

## TeaVM launcher

To test the web version of this demo, go to the Gradle window in the IDE, select `teavm/Tasks/application/runRelease`.
This will compile the code and start up a local web server. If the code compiles successfully, you will see a link to the web server among the compiler messages in the Run window:
http://localhost:8080.   Click on that link in the Run window to see the demo in a web browser.

We can also activate anti-aliasing for the web version by going into `TeaVMLauncher` and adding the following line:

```java
        config.antialiasing = true;
```

![step1browser.png](/assets/images/step1browser.png)

## Conclusions

This concludes step 1. The code we've written so far is available as release 'Step 1' in the GitHub repository.
We've set up a project with gdx-liftoff. We've learnt about the perspective camera that is used to show a 3d view.  
About models that represent the shape and texture of a 3d model and model instances that place a model
somewhere in the game world. We've learnt that the model batch is used to render model instances.
And we've seen our first camera controller. In the next step, we're going to improve on that camera controller.
