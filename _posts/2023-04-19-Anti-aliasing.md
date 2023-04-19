# Anti-aliasing (or how to kill the jaggies with one line of code)

I was watching a youtube video of JamesTKhan about [importing GLTF files into LibGDX](https://www.youtube.com/watch?v=e-3OMXY9bDU&list=PLjUR2MkQ0cuHZ70Ps8F9WMyoyKHKAbYvQ&ab_channel=JamesTKhan)
and there he mentioned in passing how to activate anti-aliasing in the desktop version.

This was a little piece of knowledge I had never came across before. It takes just a few seconds to add, literally one line of code, 
and if really improves the visual quality by gettig rid of those jagged edges. Or at least on the desktop version.


If you are running LibGDX using LWGL2 just add set config.samples to, for example, 4 to use anti-aliasing. I supposed you can put the number higher at some performance cost.

```java
public class DesktopLauncher {
	public static void main (String[] arg) {
		LwjglApplicationConfiguration config = new LwjglApplicationConfiguration();
		config.samples = 4;
		new LwjglApplication(new Main(), config);
	}
}
```

With the newer versions of LibGDX based on LWGL3 it is slightly more complicated, but still only a one-liner: use config.setBackBuffer().
The last value sets the samples to use. (The other values are bits for red, green, blue, alpha, depth and stencil respectively).

```java
public class DesktopLauncher {
	public static void main (String[] arg) {
		Lwjgl3ApplicationConfiguration config = new Lwjgl3ApplicationConfiguration();
		config.setBackBufferConfig(8, 8, 8, 8, 16, 0, 4);		// anti-aliasing 4 samples
		new Lwjgl3Application(new Main(), config);
	}
}
```

The following pictures show the before and after shots:


![samples](https://user-images.githubusercontent.com/49096535/233168129-39dda6d8-1784-4a15-b18d-2ded64bf6629.png)



Happy coding!

[1][Importing 3D Models From Blender to libGDX with gdx-gltf](https://www.youtube.com/watch?v=e-3OMXY9bDU&list=PLjUR2MkQ0cuHZ70Ps8F9WMyoyKHKAbYvQ&ab_channel=JamesTKhan)
