## Instanced Rendering (no go for HTML platform) (or is it?)

As I mentioned before, the scenery objects (trees and rocks) in the War of the Pixels game would be classic candidates for instanced rendering.  
This is a technique where you render the same mesh in many different positions, but you only need to pass the mesh once to the GPU together with a list of positions where it needs to go.
With the technique you could even add hundreds of thousands of blades of grass to the terrain.

There is an instanced rendering test [InstancedRenderingTest.java][1] as part of the LibGDX test suite.  However, it requires GL30 to run. In particular, the GL function 'glVertexAttribDivisor` is not supported until OpenGL 3.3.

It works on desktop. You may need to add the following line to DesktopLauncher.java to make sure you're running OpenGL 3.3:

```java
config.setOpenGLEmulation(Lwjgl3ApplicationConfiguration.GLEmulation.GL30, 3, 3);
```

However, the HTML/GWT platform is only running WebGL 1.0 which is roughly equivalent to GLSE 2.0.  So instanced rendering is not supported for HTM for now.

Since HTML this is the target platform for this game, I will leave this for a future project.

If you are desperate to get instanced rendering working on web pages, you could have a look at the proposed [update][2] of LibGDX to include Web GL 2.0, which is roughly equivalent to GL ES 3.0. No idea when this will be included in mainstream LibGDX. 


# Post Scriptum (September 2023)
With the release of LibGDX 1.12, the HTML/GWT platform supports WebGL2.0.  Also the HTML/TeaVM platform has experimental support for WebGL2.0. 

I have a first demo of instanced rendering on HTML [here][3]. Still requires some further work to be more robust. Performance varies per platform. For example, instancing appears to be very slow on Apple's Safari browser. Maybe my next jame game will have some instanced scenery?





[1]: https://github.com/libgdx/libgdx/blob/master/tests/gdx-tests/src/com/badlogic/gdx/tests/gles3/InstancedRenderingTest.java "InstancedRenderingTest.java"
[2]: https://github.com/libgdx/libgdx/pull/7037 "pull request to add WebGL 2.0"
[3]: https://monstrous-software.itch.io/corn-field "corn field demo"
